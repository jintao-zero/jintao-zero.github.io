---
layout: post
title: "golang创建、解压.tar.gz文件简单库"
date: 2017-01-05 19:17:06 +0800
comments: true
categories: program
tags: golang
---
golang提供了`tar`包和`compress/gzip`包进行文件打包和压缩，但是没有函数同时进行打包和压缩，下面利用打包和压缩功能，实现一个简单的制作和解压.tar.gz文件的功能。
##tar打包功能
利用tar中`Writer`和`Reader`可以实现将文件和文件夹进行打包的功能。  
###打包
创建目标文件  

	f, err := os.Create(dstPath)
	
创建`Writer`

	tw := tar.NewWriter(f)
循环遍历文件夹，写入Writer  
	
	fileInfo, err := os.Stat(path)
	if err != nil {
		return err
	}
	if fileInfo.Mode().IsRegular() {
		header, err := tar.FileInfoHeader(fileInfo, "")
		if err != nil {
			return err
		}
		header.Name = path
		if err = tw.WriteHeader(header); err != nil {
			return err
		}
		file, err := os.Open(path)
		if err != nil {
			return err
		}
		if _, err = io.Copy(tw, file); err != nil {
			return err
		}
	}

	if fileInfo.Mode().IsDir() {
		// tar each file and dir in the dir
		var file *os.File
		if file, err = os.Open(path); err != nil {
			return err
		}
		fileInfos, err := file.Readdir(0)
		if err != nil {
			return err
		}
		for _, info := range fileInfos {
			if err = tarPath(filepath.Join(path, info.Name()), tw); err != nil {
				return err
			}
		}
	}
<!-- more -->
###解包 
打开源文件

	tarFile, err := os.Open(srcPath)

创建Reader  

	tr := tar.NewReader(tarFile)

循环遍历解压文件  
	
	for {
		hdr, err := tr.Next()
		if err == io.EOF {
			break
		}
		fullPath := filepath.Join(dstPath, hdr.Name)
		os.MkdirAll(filepath.Dir(fullPath), os.ModePerm)
		log.Println("fullPath", fullPath)
		file, err := os.Create(fullPath)
		if err != nil {
			return err
		}
		if _, err := io.Copy(file, tr); err != nil {
			return err
		}
		file.Close()
	}
	
##Gzip压缩功能
###压缩
创建目标文件

	dstFile, err := os.Create(dstPath)
创建Writer
	
	gw := gzip.NewWriter(dstFile)

将源内容写入Writer
	
	if _, err = io.Copy(gw, srcFile); err != nil {
		return err
	}
###解压
打开源文件  
	
	srcFile, err := os.Open(srcPath)
创建Reader
	
	gr, err := gzip.NewReader(srcFile)
进行解压 
	
	if _, err = io.Copy(dstFile, gr); err != nil {
		return err
	}

##.tar.gz文件
将上面tar包进行打包和解包湿的的`io.Writer`和 `io.Reader`实参传入`gzip.Writer`和`gzip.Reader` ，即可将打包的目的文件进行压缩和解包时的源文件进行解压  
###打包压缩  

	func Targz(srcPath, dstPath string) error {
	dstFile, err := os.Create(dstPath)
	if err != nil {
		return err
	}
	defer dstFile.Close()
	gw := gzip.NewWriter(dstFile)
	defer gw.Close()
	tw := tar.NewWriter(gw)
	defer tw.Close()
	if err = tarPath(srcPath, tw); err != nil {
		dstFile.Close()
		os.Remove(dstPath)
		return err
	}
	return nil
}

###解压解包  
	srcFile, err := os.Open(srcPath)
	if err != nil {
		return err
	}
	gr, err := gzip.NewReader(srcFile)
	if err != nil {
		return err
	}
	tr := tar.NewReader(gr)
	for {
		hdr, err := tr.Next()
		if err == io.EOF {
			break
		}
		fullPath := filepath.Join(dstPath, hdr.Name)
		os.MkdirAll(filepath.Dir(fullPath), os.ModePerm)
		file, err := os.Create(fullPath)
		if err != nil {
			return err
		}
		if _, err := io.Copy(file, tr); err != nil {
			return err
		}
		file.Close()
	}

###备注  
完整源代码请参考[targz](https://github.com/jintao-zero/targz)

	